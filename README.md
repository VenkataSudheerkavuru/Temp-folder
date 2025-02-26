<!-- Button to Open the Modal -->
<button class="btn btn-primary" data-bs-toggle="modal" data-bs-target="#formModal">
  Open Form Modal
</button>

<!-- Modal Structure -->
<div class="modal fade" id="formModal" tabindex="-1" aria-labelledby="formModalLabel" aria-hidden="true">
  <div class="modal-dialog">
    <div class="modal-content">
      <!-- Modal Header -->
      <div class="modal-header">
        <h5 class="modal-title" id="formModalLabel">Fill Out the Form</h5>
        <button type="button" class="btn-close" data-bs-dismiss="modal" aria-label="Close"></button>
      </div>

      <!-- Modal Body (Form) -->
      <div class="modal-body">
        <form [formGroup]="form" (ngSubmit)="onSubmit()">
          <!-- Name Field -->
          <div class="mb-3">
            <label for="name" class="form-label">Name</label>
            <input type="text" id="name" class="form-control" formControlName="name" />
          </div>

          <!-- Email Field -->
          <div class="mb-3">
            <label for="email" class="form-label">Email</label>
            <input type="email" id="email" class="form-control" formControlName="email" />
          </div>
        </form>
      </div>

      <!-- Modal Footer -->
      <div class="modal-footer">
        <button type="button" class="btn btn-secondary" data-bs-dismiss="modal">Close</button>
        <button type="submit" class="btn btn-primary" [disabled]="form.invalid" (click)="onSubmit()">Submit</button>
      </div>
    </div>
  </div>
</div>
